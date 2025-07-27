# IMAGE_SIZE_COMPOSITE.md

This document explains how ExifTool implements the Composite:ImageSize tag.

## Overview

The ImageSize composite tag provides a formatted string representation of image dimensions in the format "width x height" (e.g., "1920 x 1080").

## Implementation Details

### Location

The ImageSize composite tag is defined in `lib/Image/ExifTool/Exif.pm` at line 4641.

### Tag Definition

```perl
ImageSize => {
    Require => {
        0 => 'ImageWidth',
        1 => 'ImageHeight',
    },
    Desire => {
        2 => 'ExifImageWidth',
        3 => 'ExifImageHeight',
        4 => 'RawImageCroppedSize', # (FujiFilm RAF images)
    },
    # use ExifImageWidth/Height only for Canon and Phase One TIFF-base RAW images
    ValueConv => q{
        return $val[4] if $val[4];
        return "$val[2] $val[3]" if $val[2] and $val[3] and
                $$self{TIFF_TYPE} =~ /^(CR2|Canon 1D RAW|IIQ|EIP)$/;
        return "$val[0] $val[1]" if IsFloat($val[0]) and IsFloat($val[1]);
        return undef;
    },
    PrintConv => '$val =~ tr/ /x/; $val',
}
```

### Dependencies

**Required tags:**
- `ImageWidth` (index 0)
- `ImageHeight` (index 1)

**Desired tags (optional):**
- `ExifImageWidth` (index 2)
- `ExifImageHeight` (index 3)
- `RawImageCroppedSize` (index 4) - Used for FujiFilm RAF images

### Value Conversion Algorithm

The ValueConv follows this precedence order:

1. **RawImageCroppedSize**: If available (FujiFilm RAF), use it directly
2. **ExifImageWidth/Height**: Use for specific Canon and Phase One TIFF-based RAW formats:
   - CR2 (Canon Raw 2)
   - Canon 1D RAW
   - IIQ (Phase One)
   - EIP (Phase One Enhanced Image Package)
3. **ImageWidth/Height**: Use standard dimensions if both are valid numbers
4. **Return undef**: If no valid dimensions found

### Validation

The `IsFloat()` function (ExifTool.pm:5850) validates numeric values:
- Accepts integers and floating point numbers
- Allows scientific notation (e.g., "1.5E3")
- Supports comma as decimal separator for other locales (converts to period)

### Print Conversion

The PrintConv formatting:
```perl
$val =~ tr/ /x/; $val
```

This replaces the space separator with 'x', converting:
- Input: "1920 1080" 
- Output: "1920x1080"

## Special Considerations

### RAW Format Handling

For Canon and Phase One TIFF-based RAW images, ExifTool prefers ExifImageWidth/Height over the standard ImageWidth/Height. This is because these RAW formats may have different dimensions in their main IFD versus their EXIF IFD.

### FujiFilm RAF Support

FujiFilm RAF images can provide a pre-formatted RawImageCroppedSize value that takes precedence over calculating from width/height.

### Edge Cases

1. **Missing Required Tags**: Returns undef if ImageWidth or ImageHeight are missing
2. **Non-numeric Values**: The IsFloat validation ensures only valid numbers are accepted
3. **Partial Data**: Won't use ExifImageWidth/Height unless both are present
4. **Unknown TIFF Types**: Falls back to standard ImageWidth/Height for non-Canon/Phase One formats

## Related Tags

- **Megapixels**: Composite tag that depends on ImageSize, calculates total pixels
- **ImageWidth/Height**: Standard EXIF/TIFF dimensions
- **ExifImageWidth/Height**: Dimensions from EXIF IFD (may differ for some formats)

## Common Pattern

This PrintConv pattern (`tr/ /x/`) is used consistently across ExifTool for dimension formatting:
- MIE module: Various size tags
- Sony: Preview image sizes
- Pentax: Focus frame size
- QuickTime: Video dimensions
- And many others

The pattern ensures consistent "widthxheight" formatting across all dimension-related tags.