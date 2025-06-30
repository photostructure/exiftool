# ExifTool ValueConv Deep Dive

## Overview

ValueConv (Value Conversion) is ExifTool's core system for converting raw metadata values into logical, usable forms. Unlike PrintConv which makes data human-readable, ValueConv transforms raw binary/encoded data into programmatically useful values while preserving data integrity for round-trip operations.

## Core Architecture

ValueConv operates in the middle stage of tag processing:

1. **Raw Data** → Format conversion
2. **Formatted Data** → **ValueConv** → **Logical Value**
3. **Logical Value** → PrintConv → Display String

**Key differences from PrintConv:**

- ValueConv preserves data types and precision
- Required for mathematical operations and data analysis
- Applied before PrintConv in the processing chain
- Critical for write operations through ValueConvInv

## ValueConv Implementation Types

**Complete documentation:** See README:610-664 for comprehensive ValueConv specifications.

### 1. Hash Reference Lookups (`ValueConv => \%hash`)

**From README:617-630:** Hash references provide lookup tables with special key support.

```perl
# Simple lookup
%canonFlashMode = (
    0 => 'Off',
    1 => 'On',
    2 => 'Auto',
);
ValueConv => \%canonFlashMode,
```

**Special Hash Keys (README:621-623):**

- **BITMASK**: Reference to hash for decoding individual bits
- **OTHER**: Subroutine for unknown values (takes value, inverse_flag, conv_hash)
- **Notes**: Documentation (used with SeparateTable flag)

### 2. Code String Evaluation (`ValueConv => 'expression'`)

**From README:630-634:** Perl expressions evaluated with special variables.

```perl
# Mathematical conversion
ValueConv => '$val / 100',

# Conditional logic
ValueConv => '$val ? 1/$val : 0',

# Complex calculations
ValueConv => 'exp(Image::ExifTool::Canon::CanonEv($val)*log(2)/2)',
```

**Available Variables (README:630-632):**

- `$val` - the raw value
- `$self` - ExifTool object reference
- `$tag` - the tag key
- `@val`, `@prt`, `@raw` - arrays for Composite tags

### 3. Code Reference (`ValueConv => \&subroutine` or `ValueConv => sub { ... }`)

**From README:631-632:** Direct function calls for complex processing.

```perl
# Anonymous subroutine
ValueConv => sub {
    my ($val, $et) = @_;
    return '<err>' if length($val) < 4;
    my $len = unpack('N', $val) * 2;
    return $et->Decode(substr($val, 4, $len), 'UCS2', 'MM');
},
```

### 4. Array Reference (`ValueConv => [\&conv1, \&conv2, ...]`)

**From README:614-617:** Different conversions for array elements.

```perl
# Canon.pm:2028 - Multiple picture style conversions
ValueConv => [\%pictureStyles, \%pictureStyles, \%pictureStyles],
```

**Special Value:** `'REPEAT'` repeats previous conversion for remaining elements.

### 5. Binary Data Return Pattern (`ValueConv => '\$val'`)

**From README:355-356 (Binary flag):** Returns scalar reference for binary data.

```perl
# Return binary data unchanged
ValueConv => '\$val',     # Equivalent to Binary => 1
ValueConvInv => '$val',
```

## ValueConvInv (Inverse Conversion)

**Complete documentation:** See README:690-703 for full ValueConvInv specifications.

**Purpose:** Converts logical values back to raw format for writing operations.

```perl
# Forward and inverse pair
ValueConv    => '$val / 100',
ValueConvInv => '$val * 100',
```

**From README:695-698:** Error handling via `warn()` or empty warning (`"\n"`).

**From README:692-694:** DataMember tags NOT available in inverse conversions - use Condition or RawConvInv instead.

## Processing Logic and Location

**Primary Function:** `GetValue()` in ExifTool.pm:3378-3667  
**Conversion Loop:** Lines 3460-3627

**Key Processing Differences:**

- **Hash lookups:** Direct value mapping with BITMASK/OTHER support
- **Code strings:** `eval` with variable context
- **Code references:** Direct subroutine calls
- **Binary protection:** Skips SCALAR refs unless ConvertBinary flag set

**From README:637-639:** ValueConv evaluated on-demand when tag values requested, allowing access to other tag values via `$self->GetValue("Tag","Raw")`.

## Format-Specific Patterns and Edge Cases

### Dynamic Module Loading

```perl
# XMP.pm:1257 - Load PDF module on demand
ValueConv => 'require Image::ExifTool::PDF; Image::ExifTool::PDF::ConvertPDFDate($val)',
```

### Self-Reference Patterns

```perl
# Access ExifTool object members
ValueConv => '$val / ($$self{FocalUnits} || 1)',
```

### Encryption/Decryption

```perl
# Pentax.pm:745-751 - Decrypt by bit toggling
ValueConv => sub {
    my $val = shift;
    return $val unless length($val) == 4;
    my @a = map { $_ ^ 0xff } unpack("C*",$val);  # Toggle all bits
    return sprintf('%d %.2d %.2d %.2d', @a);
},
```

### Complex Binary Parsing

```perl
# Photoshop.pm:230-245 - Unicode string arrays
ValueConv => sub {
    my ($val, $et) = @_;
    return '<err>' if length($val) < 4;
    my $num = unpack('N', $val);
    my ($i, @vals);
    # ... complex parsing logic
    return \@vals;
},
```

## Surprising and "Tribal Knowledge" Discoveries

### 1. Binary vs. Logical Data Boundaries

**From README:641-644:** ValueConv may return SCALAR references for binary data, but these are NOT processed by PrintConv unless ConvertBinary flag is set.

```perl
# Canon.pm:1141 - Conditional binary return
ValueConv => 'length($val) > 64 ? \$val : $val',
```

### 2. Composite Tag Array Processing

**From README:653-657:** Composite tags access source values via `@val`, `@prt`, `@raw` arrays corresponding to Require/Desire indices.

```perl
# Apple.pm:356 - Ratio calculation from array
ValueConv => '$val[1] ? $val[0] / $val[1] : undef',
```

### 3. Error Handling Philosophy

**From README:645-651:** ValueConv should always return defined values - use RawConv for validity testing that may return undef.

### 4. Module Dependency Management

Dynamic `require` statements handle optional dependencies:

```perl
ValueConv => 'require Image::ExifTool::XMP; Image::ExifTool::XMP::ConvertXMPDate($val)',
```

### 5. Evaluation Context Richness

**From README:634:** ValueConv expressions have access to the advanced formatting expression via `$$self{FMT_EXPR}`.

### 6. Write-Time Validation Complexity

**From README:692-694:** DataMember restrictions in ValueConvInv create asymmetric complexity - forward conversion can access other tags, but inverse cannot.

### 7. Performance Optimization

**From README:639-641:** ValueConv is evaluated on-demand, only when tag values are requested, unlike RawConv which executes for all extracted tags.

## Key Architectural Insights

### Processing Order Dependencies

**From README:669-673:** ValueConv operates on format-converted data, not raw binary data. This creates a three-stage pipeline that must be understood for complex conversions.

### Write Operation Asymmetry

Forward operations can access the full ExifTool state and other tag values, but inverse operations are limited to avoid circular dependencies during the writing process.

### Binary Data Handling

The distinction between binary (SCALAR ref) and logical data affects the entire processing pipeline and determines whether PrintConv is applied.

## Best Practices Summary

**Based on README specifications:**

1. **Use appropriate type:** Hash for lookups, expressions for math, code refs for complex logic
2. **Provide ValueConvInv:** Required for writable tags (README:690-691)
3. **Handle edge cases:** Use OTHER functions for unknown values
4. **Consider performance:** ValueConv is on-demand, so complex operations are acceptable
5. **Maintain data integrity:** ValueConv should preserve precision for round-trip operations
6. **Use RawConv for validation:** Don't return undef from ValueConv (README:647-651)

## Conclusion

ValueConv represents the logical processing heart of ExifTool's metadata system. Unlike PrintConv's focus on human readability, ValueConv transforms raw data into programmatically useful forms while maintaining the precision and data types necessary for mathematical operations and write operations.

The system's flexibility allows handling everything from simple mathematical conversions to complex binary parsing, encryption/decryption, and dynamic module loading, making it essential for ExifTool's ability to handle hundreds of different metadata formats.
