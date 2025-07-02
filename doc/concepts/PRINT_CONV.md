# ExifTool PrintConv Deep Dive

## Overview

The `PrintConv` (Print Conversion) feature converts tag values to human-readable form. For complete technical details, see `lib/Image/ExifTool/README` lines 665-677.

**Processing flow:** Raw Data → ValueConv → **PrintConv** → Display String

**Key points from README:**

- PrintConv operates on ValueConv results, not raw values
- Only applied if PrintConv option enabled and ValueConv result isn't a scalar reference
- List values joined with '; ' separator

## PrintConv Types and Patterns

### PrintConv Format Types

**Basic formats covered in README:665-677:**

- Hash reference lookup
- Scalar Perl expression
- Code reference (subroutine)
- List reference (see README:615-617 for REPEAT feature)

**Expression variables:** `$self`, `$val`, `$tag` - see README:669-670

### 1. Multi-line Code Blocks (`PrintConv => q{ ... }`)

Extended Perl code for complex processing.

```perl
# Example from Canon.pm:1615 - Firmware revision parsing
PrintConv => q{
    my $rev = sprintf("%.8x", $val);
    my ($rel, $v1, $v2, $r1, $r2) = ($rev =~ /^(.)(.)(..)0?(.+)(..)$/);
    my %r = ( a => 'Alpha ', b => 'Beta ', '0' => '' );
    $rel = defined $r{$rel} ? $r{$rel} : "Unknown($rel) ";
    return "$rel$v1.$v2 rev $r1.$r2",
},
```

**Usage:** Multi-step parsing, regex processing, complex algorithms

### 2. BITMASK Conversion (`PrintConv => { BITMASK => { ... } }`)

**Technical details:** See README:621-622 for BITMASK hash functionality  
**Configuration:** BitsPerWord/BitsTotal covered in README:928-932

```perl
# Example from Canon.pm:1640
PrintConv => {
    0 => '(none)',
    BITMASK => {
        0 => 'People',   1 => 'Scenery',   2 => 'Events',
        3 => 'User 1',   4 => 'User 2',    5 => 'User 3',   6 => 'To Do',
    },
},
```

**Output:** Comma-separated list: "People, Events, User 1"

### 3. OTHER Function Pattern (`OTHER => sub { ... }`)

**Technical details:** See README:623-626 for OTHER subroutine functionality

```perl
# Example - Canon's humorous 0xdeadbeef handling
my %psConv = (
    -559038737 => 'n/a', # = 0xdeadbeef ! LOL
    OTHER => sub { shift },
);
```

**Signature:** `OTHER => sub { my ($val, $inv, $conv) = @_; ... }`

## PrintConvInv (Inverse Conversion)

**See README:705-708** for complete PrintConvInv documentation.

**Key points:**

- Required for writable tags with PrintConv (unless WriteAlso used)
- Variables: `$val`, `$self`, `$wantGroup`
- Error handling via warn() or empty warning ("\n")

**Example - Canon firmware parsing inverse:**

```perl
PrintConvInv => q{
    $_=$val; s/Alpha ?/a/i; s/Beta ?/b/i;
    s/Unknown ?\((.)\)/$1/i; s/ ?rev ?(.)\./0$1/; s/ ?rev ?//;
    tr/a-fA-F0-9//dc; return hex $_;
},
```

## Special Features and Modifiers

**Most PrintConv modifiers covered in README:**

- **PrintConvColumns** (README:710-712)
- **PRINT_CONV** table default (README:144-146)
- **PrintHex** (README:482-486)
- **BitsPerWord/BitsTotal** (README:928-932)
- **PrintSort** (README:492-493)
- **SeparateTable** (README:529-533)

**Key usage:**

```perl
PrintConvColumns => 2,  # Column count for docs
PrintHex => 1,          # Show hex for unknown values
BitsPerWord => 8,       # For BITMASK tags
```

## Language Localization

PrintConv values can be localized through language files in `lib/Image/ExifTool/Lang/*.pm`.

```perl
# Example from Lang/de.pm - German translations
'AEBAutoCancel' => {
   Description => 'Automatisches Bracketingende',
   PrintConv => { 'Off' => 'Aus', 'On' => 'Ein' },
},
```

## Odd, Surprising, and "Tribal Knowledge" Discoveries

### 1. The 0xdeadbeef Easter Egg

```perl
# Canon.pm:1130 - Picture style conversion
my %psConv = (
    -559038737 => 'n/a', # = 0xdeadbeef ! LOL
    OTHER => sub { shift },
);
```

The comment reveals that -559038737 is actually 0xdeadbeef (a famous hex "magic number" in programming). This is a humorous Easter egg showing that Canon sometimes uses this classic debugging value.

### 2. Complex Firmware Parsing Magic

The firmware revision parser (Canon.pm:1615) contains sophisticated reverse-engineering:

```perl
# Decodes: 0xAVVVRR00 format where:
#  A = 'a' for alpha, 'b' for beta?
#  V = version? (100,101 for normal releases, 100,110,120,130,170 for alpha/beta)
#  R = revision? (01-07, except 00 for alpha/beta releases)
```

This represents deep reverse-engineering of Canon's internal firmware numbering scheme.

### 3. Byte-Order Weirdness

```perl
# Canon.pm:1161 - Focus distance with odd byte ordering
my %focusDistanceByteSwap = (
    # this is very odd (little-endian number on odd boundary),
    # but it does seem to work better with my sample images - PH
    Format => 'int16uRev',
    # ...
);
```

This reveals Canon's inconsistent byte ordering in some fields, requiring special handling.

### 4. The "n/a" vs undef Pattern

Many PrintConv definitions use `'n/a'` strings instead of undef for missing values:

```perl
0x7fff => 'n/a',    # Canon.pm:2625
0xffff => 'n/a',    # Canon.pm:6520
```

This suggests camera firmware explicitly sets these "not available" sentinel values rather than leaving fields empty.

### 5. Evaluation Context Variables and Processing Order

The eval'd PrintConv strings have access to several variables beyond `$val`:

- `@val` - array of values for multi-value tags
- `@prt` - array of print-converted values
- `@raw` - array of raw unconverted values
- `$self` - ExifTool object
- `$tag` - the tag key

**Critical Processing Detail from README:674-675:** "Note that the print conversion is only done if the PrintConv option is enabled (which it is by default), and if the result of the ValueConv is not a scalar reference."

**List Handling from README:676-677:** "If it is a list reference, then the converted values are joined by '; ' in the output string."

This allows extremely sophisticated conversions that can reference other tag values or ExifTool state.

### 6. BITMASK Zero-Value Special Handling

```perl
PrintConv => {
    0 => '(none)',        # Special case for zero
    BITMASK => { ... },
},
```

When a bitmask value is 0 (no bits set), it displays "(none)" rather than an empty string. This pattern appears consistently across different modules.

### 7. Inverse Conversion Complexity

Some PrintConvInv functions are more complex than their forward counterparts:

```perl
# Forward: simple sprintf
PrintConv => 'sprintf("%.4x%.5d",$val>>16,$val&0xffff)',

# Inverse: complex parsing
PrintConvInv => sub {
    # Complex regex and parsing logic...
},
```

This reflects the asymmetric complexity of parsing vs. formatting.

## Implementation Notes

**Core processing details covered in README:665-677**

**Key implementation points:**

- Processing location: ExifTool.pm:3542-3596
- PrintConv operates on ValueConv results, not raw values
- Done "on demand" - may occur after tag extraction
- Binary data excluded unless ConvertBinary flag set
- Failures fallback to "Unknown (value)" format

**When PrintConv is skipped:**

1. PrintConv option disabled
2. ValueConv result is scalar reference (binary data)
3. Tag has `PrintConv => undef`

## Best Practices

1. **Use hash references** for simple lookups
2. **Use OTHER functions** for handling unknown values gracefully
3. **Include PrintConvInv** for writable tags
4. **Add comments** explaining complex conversion logic
5. **Consider localization** for user-facing strings
6. **Test edge cases** including zero, negative, and extreme values

## JSON Output and Numeric PrintConv Values

### Important Discovery: PrintConv Can Return Numeric Values

While PrintConv traditionally formats values for human readability, some PrintConv functions return numeric strings that become JSON numbers:

**Examples**:
- `FNumber`: PrintConv returns `"4.0"` → JSON outputs as `4.0` (number)
- `ExposureTime`: PrintConv returns `"1/2000"` → JSON outputs as `"1/2000"` (string)
- `FocalLength`: PrintConv returns `"24.0 mm"` → JSON outputs as `"24.0 mm"` (string)

### The FNumber Case Study

```perl
# From Exif.pm
sub PrintFNumber($)
{
    my $val = shift;
    if (Image::ExifTool::IsFloat($val) and $val > 0) {
        # round to 1 decimal place, or 2 for values < 1.0
        $val = sprintf(($val<1 ? "%.2f" : "%.1f"), $val);
    }
    return $val;  # Returns numeric value, not string!
}
```

**Key Point**: PrintFNumber formats the number but returns it as a numeric value, not a string with "f/" prefix. This allows JSON to encode it as a number.

### Implementation Guidance for exif-oxide

1. **Preserve ExifTool's Exact Behavior**: Some PrintConv functions format numeric values but keep them numeric for JSON encoding
2. **Don't Assume PrintConv = String**: The name is misleading - it's more "DisplayConv" that can return any type
3. **Match Quirks Exactly**: Including "24.0 mm" with the unnecessary `.0` for whole numbers

### The -# Flag Behavior

ExifTool's `-TagName#` syntax:
- Disables PrintConv for that specific tag
- Shows the ValueConv result (or raw value if no ValueConv)
- In JSON, maintains proper type (numbers unquoted, strings quoted)

```bash
# Normal output
exiftool -j image.jpg
{"FNumber": 4.0}  # PrintConv result as number

# With -# flag
exiftool -j -FNumber# image.jpg  
{"FNumber": 4}    # ValueConv/raw result
```

## Reference

For complete technical documentation on PrintConv and all tag information hash entries, see `lib/Image/ExifTool/README` lines 665-677 and the full tag information specification starting at line 280.
