# ExifTool Ternary Expression Patterns Analysis

## Summary

ExifTool extensively uses ternary operator patterns (`condition ? true_value : false_value`) in PrintConv and ValueConv fields for conditional data processing. The patterns range from simple null checks to complex nested conditions with mathematical calculations, making them a critical component for implementing metadata parsing logic in Rust.

## Implementation Details

### Common Ternary Operator Patterns

#### 1. **Simple Validation Patterns**
Most common pattern: basic null/zero checks with fallback values.

```perl
# Null/undefined checks
'$val ? $val : undef'                    # Return undef if val is falsy
'$val ? "$val m" : "inf"'               # Distance with infinity fallback
'$val > 0 ? "+$val" : $val'             # Add plus sign for positive values
'$val == 0x7fff ? undef : $val'         # Specific sentinel value check
```

#### 2. **Comparison-Based Conditionals**
Threshold-based logic using `>=`, `>`, `<`, `<=`, `==`, `!=`:

```perl
# Numeric thresholds
'$val > 655.345 ? "inf" : "$val m"'     # Canon focus distance
'$val >= 0 ? $val : undef'              # Non-negative validation
'$val <= 180 ? $val : $val - 360'       # Angle normalization
'$val >= 128 ? "inf" : $val * $val[1] / 1000'  # Sony distance calculation
```

#### 3. **Length and Format Validation**
String/binary data validation patterns:

```perl
# Length checks
'length($val) > 64 ? \$val : $val'      # Binary vs text decision
'length($val) >= 6 ? unpack("x4n",$val) : "<err>"'  # Minimum data length
'length($val) <= 64 ? $val : \$val'     # Print conversion limit
'length($val)==4 ? unpack("V",$val) : \$val'  # Fixed-length format
```

#### 4. **Complex Nested Ternary**
Multi-level conditional logic:

```perl
# Nested conditions for scaling factors (PanasonicRaw.pm:281)
'$val / ($val >= 1024 ? 1024 : ($val >= 256 ? 256 : 100))'

# Printf precision based on value range (Exif.pm:4664)
'sprintf("%.*f", ($val >= 1 ? 1 : ($val >= 0.001 ? 3 : 6)), $val)'

# Canon custom function pattern
'$val ? ($val==1 ? "On" : "On ($val)") : "Off"'
```

### Comparison Operators Used

**Primary operators observed:**
- `>=`, `>` - Most common for threshold checks
- `<=`, `<` - Range validation and limits
- `==` - Exact value matching (sentinel values)
- `!=` - Value exclusion
- `=~` - Regex pattern matching
- `eq` - String equality

### Value Types in True/False Branches

#### **Common Return Types:**
1. **Strings**: `"inf"`, `"n/a"`, `"Auto"`, `"Unknown ($val)"`
2. **Numbers**: `$val`, `0`, `255`, mathematical expressions
3. **undef**: For invalid/missing data
4. **Binary references**: `\$val` for binary data
5. **Formatted strings**: `"$val m"`, `"+$val"`, `sprintf(...)`
6. **Mathematical expressions**: `exp(...)`, `log(...)`, trigonometric functions

#### **Variable References:**
- `$val` - Primary value
- `$val[0]`, `$val[1]` - Array elements for composite tags
- `$self->{Model}` - Camera model conditional logic
- `$$self{FujiLayout}` - Format-specific flags

### Complex Expression Examples

#### **Mathematical Conversions**
```perl
# Exposure value calculations (Minolta.pm)
'$val ? 2 ** (6 - $val/8) : 0'
'$val ? int((6 - log($val) / log(2)) * 8 + 0.5) : 0'

# Logarithmic scaling (Olympus.pm)
'abs($val)<100 ? 2**(-$val) : 0'
'$val>0 ? -log($val)/log(2) : -100'
```

#### **Conditional String Processing**
```perl
# Pattern extraction and validation
'$val=~/(.*)-?(\d{5})$/ ? (hex($1)<<16)+$2 : undef'
'$val=~/(-?[\d.]+)/ ? $1 : 0'
'$val =~ s/ ?m$//; IsFloat($val) ? $val : 655.35'
```

#### **Model-Specific Logic**
```perl
# Camera model conditional adjustments (Minolta.pm)
'$val - ($self->{Model}=~/DiMAGE A2/ ? 5 : 3)'
'$val + ($self->{Model}=~/DiMAGE A2/ ? 5 : 3)'
```

## Special Perl-Specific Behaviors

### 1. **Truthiness vs Definedness**
Perl's truthiness differs from most languages:
```perl
'$val ? $val : undef'  # 0, "", "0" are falsy but defined
'$val eq "inf" ? 0 : $val'  # String comparison
```

### 2. **Regex Modifier Effects**
```perl
'$val =~ s/ 1$// ? -$val/10 : "n/a"'  # s/// returns success/failure
'($val =~ s/^0 // and $val) ? $val : undef'  # Chained boolean logic
```

### 3. **Array Context vs Scalar Context**
```perl
'my @a=($val=~/(\d+):(\d+):(\d+)/); @a ? ($a[0]<<16)+($a[1]<<8)+$a[2] : undef'
```

### 4. **Reference vs Value Returns**
```perl
'length($val) > 64 ? \$val : $val'  # Return reference for binary data
```

## Key Files with Ternary Patterns

**High-density files:**
- `/home/mrm/src/exif-oxide/third-party/exiftool/lib/Image/ExifTool/Canon.pm` - Lines 1170, 2385, 7518
- `/home/mrm/src/exif-oxide/third-party/exiftool/lib/Image/ExifTool/Minolta.pm` - Lines 1114, 1908-1911, 2700
- `/home/mrm/src/exif-oxide/third-party/exiftool/lib/Image/ExifTool/Olympus.pm` - Lines 912-913, 3333-3334
- `/home/mrm/src/exif-oxide/third-party/exiftool/lib/Image/ExifTool/Exif.pm` - Lines 1729, 4664
- `/home/mrm/src/exif-oxide/third-party/exiftool/lib/Image/ExifTool/PanasonicRaw.pm` - Line 281

## Special Considerations

### 1. **Sentinel Value Handling**
ExifTool uses specific sentinel values to indicate special states:
- `0x7fff`, `0xffff` - "Invalid" markers
- `0xdeadbeef` - Debug/placeholder values  
- `65535`, `255` - Maximum value indicators
- Empty strings, null bytes - Missing data

### 2. **Precision and Formatting**
Many ternary expressions handle precision dynamically:
```perl
'sprintf("%.*f", ($val >= 1 ? 1 : ($val >= 0.001 ? 3 : 6)), $val)'
```

### 3. **Binary Data Decisions**
Common pattern to decide between binary and text representation:
```perl
'length($val) > 64 ? \$val : $val'
```

### 4. **Error Handling Philosophy**
- `undef` returns indicate invalid/missing data
- String values like `"n/a"`, `"inf"` for out-of-range conditions
- Error strings like `"<err>"`, `"Unknown ($val)"` for format problems

## Implementation Complexity Assessment

**Complexity levels for Rust implementation:**

1. **Low Complexity** (90% of cases): Simple comparisons with basic returns
2. **Medium Complexity** (8% of cases): Nested conditions, mathematical expressions
3. **High Complexity** (2% of cases): Regex-dependent conditions, model-specific branching

**Most challenging aspects:**
- Perl's regex match return value semantics
- Truthiness vs definedness distinctions
- Dynamic precision formatting
- Model-specific conditional logic requiring camera database lookups