# APEX Conversions in ExifTool

## Overview

APEX (Additive System of Photographic Exposure) is a logarithmic system for representing photographic exposure values. ExifTool stores certain EXIF tags like ApertureValue and MaxApertureValue as APEX values, but displays them as familiar F-numbers through ValueConv and PrintConv transformations.

## Key APEX Tags

### ApertureValue (0x9202) and MaxApertureValue (0x9205)

**Location**: `lib/Image/ExifTool/Exif.pm:2317-2325` and `lines 2340-2349`

Both tags use identical conversion formulas:

```perl
# Tag definition for ApertureValue (0x9202)
0x9202 => {
    Name => 'ApertureValue',
    Notes => 'displayed as an F number, but stored as an APEX value',
    Writable => 'rational64u',
    ValueConv => '2 ** ($val / 2)',
    ValueConvInv => '$val>0 ? 2*log($val)/log(2) : 0',
    PrintConv => 'sprintf("%.1f",$val)',
    PrintConvInv => '$val',
},
```

## Conversion Formula Details

### APEX to F-Number Conversion

The APEX system defines aperture value (Av) such that:
- **F-number = 2^(Av/2)**

ExifTool implements this exactly in ValueConv:
```perl
ValueConv => '2 ** ($val / 2)'
```

For example:
- APEX value 6 → F-number = 2^(6/2) = 2^3 = 8
- APEX value 4 → F-number = 2^(4/2) = 2^2 = 4
- APEX value 3 → F-number = 2^(3/2) = 2^1.5 ≈ 2.83

### F-Number to APEX Conversion (Inverse)

The inverse formula converts F-numbers back to APEX:
- **Av = 2 × log₂(F-number)**

ExifTool implements this as:
```perl
ValueConvInv => '$val>0 ? 2*log($val)/log(2) : 0'
```

Note: `log($val)/log(2)` is the Perl idiom for log₂($val)

### PrintConv Formatting

After ValueConv converts APEX to F-number, PrintConv formats it for display:
```perl
PrintConv => 'sprintf("%.1f",$val)'
```

This formats to 1 decimal place (e.g., "8.0", "2.8", "1.4").

## Special Considerations

### FNumber vs ApertureValue

ExifTool also has an FNumber tag (0x829d) that stores the F-number directly (not as APEX). It uses a different PrintConv:

```perl
# From Exif.pm:5607-5617
sub PrintFNumber($)
{
    my $val = shift;
    if (Image::ExifTool::IsFloat($val) and $val > 0) {
        # round to 1 decimal place, or 2 for values < 1.0
        $val = sprintf(($val<1 ? "%.2f" : "%.1f"), $val);
    }
    return $val;
}
```

Key difference: FNumber uses 2 decimal places for values < 1.0 (e.g., "0.95"), while ApertureValue always uses 1 decimal place.

### MaxAperture Handling in Lens Identification

From `Exif.pm:5803-5804`:
```perl
# use MaxApertureValue if MaxAperture is not available
$maxAperture = $maxApertureValue unless $maxAperture;
```

When identifying lenses, ExifTool treats MaxAperture and MaxApertureValue as interchangeable after conversion.

## Common Issues and Debugging

### Factor of 2 Difference

If you see ApertureValue showing 16.0 when expecting 8.0, the APEX conversion is likely missing:
- APEX value 6 without conversion = 6
- APEX value 6 with correct conversion = 2^(6/2) = 8

### String vs Numeric Output

If MaxApertureValue shows as a string like "3.2" instead of numeric 3.4:
1. Check that ValueConv is being applied (converts APEX to F-number)
2. Verify PrintConv formatting is consistent
3. Consider floating-point precision in the APEX calculation

### Precision and Rounding

The APEX system can introduce small rounding differences:
- APEX for F/3.4 = 2 × log₂(3.4) ≈ 3.513
- Converting back: 2^(3.513/2) ≈ 3.396 (rounds to 3.4)
- Small differences like 3.2 vs 3.4 may indicate precision loss in the APEX storage

## Implementation Notes for exif-oxide

When implementing APEX conversions in Rust:

1. **Use exact formula**: `f_number = 2.0_f64.powf(apex_value / 2.0)`
2. **Handle edge cases**: Check for zero/negative values in inverse conversion
3. **Match PrintConv formatting**: Use consistent decimal places (1 for ApertureValue)
4. **Consider precision**: APEX values are typically stored as rationals, preserve precision through conversion
5. **Test with known values**: 
   - APEX 0 → F/1.0
   - APEX 2 → F/2.0
   - APEX 4 → F/4.0
   - APEX 6 → F/8.0