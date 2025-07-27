# Lens Tag Handling in ExifTool

**ExifTool Version:** 13.26  
**Document Version:** 1.0  
**Last Updated:** 2025-07-26

## Overview

ExifTool handles lens information through multiple EXIF tags introduced in EXIF 2.3/2.32. These tags provide standardized ways to store lens specifications and identification across camera manufacturers.

## Primary Lens Tags

### LensInfo (0xA432)

**Definition:** `Exif.pm:2922-2933`

- **Name:** LensInfo (also called LensSpecification in EXIF spec)
- **Format:** rational64u[4] - Array of 4 rational values
- **Values:** [MinFocalLength, MaxFocalLength, MinFNumberAtMinFocal, MinFNumberAtMaxFocal]
- **PrintConv:** `PrintLensInfo()` function formats as human-readable string
- **PrintConvInv:** `ConvertLensInfo()` function parses from string format

**Example Values:**
- Fixed lens: `[50, 50, 1.4, 1.4]` → `"50mm f/1.4"`
- Zoom lens: `[24, 70, 2.8, 2.8]` → `"24-70mm f/2.8"`
- Variable aperture: `[18, 55, 3.5, 5.6]` → `"18-55mm f/3.5-5.6"`

### LensMake (0xA433)

**Definition:** `Exif.pm:2934`

- **Name:** LensMake
- **Format:** string (ASCII)
- **Writable:** yes
- **Purpose:** Lens manufacturer name (e.g., "Canon", "Sigma", "Tamron")

### LensModel (0xA434)

**Definition:** `Exif.pm:2935`

- **Name:** LensModel  
- **Format:** string (ASCII)
- **Writable:** yes
- **Purpose:** Full lens model name (e.g., "Canon EF 24-70mm f/2.8L II USM")

### LensSerialNumber (0xA435)

**Definition:** `Exif.pm:2936`

- **Name:** LensSerialNumber
- **Format:** string (ASCII)
- **Writable:** yes
- **Purpose:** Unique lens serial number

## PrintConv Implementation

### PrintLensInfo Function

**Location:** `Exif.pm:5694-5712`

The `PrintLensInfo()` function converts the 4 rational values into a human-readable format:

```perl
sub PrintLensInfo($)
{
    my $val = shift;
    my @vals = split ' ', $val;
    return $val unless @vals == 4;
    
    # Validate all 4 values are numeric or special values
    my $c = 0;
    foreach (@vals) {
        Image::ExifTool::IsFloat($_) and ++$c, next;
        $_ eq 'inf' and $_ = '?', ++$c, next;
        $_ eq 'undef' and $_ = '?', ++$c, next;
    }
    return $val unless $c == 4;
    
    # Format: MinFocal[-MaxFocal]mm f/MinF[-MaxF]
    $val = $vals[0];
    # Pentax Q writes zero for upper value of fixed-focal-length lenses
    $val .= "-$vals[1]" if $vals[1] and $vals[1] ne $vals[0];
    $val .= "mm f/$vals[2]";
    $val .= "-$vals[3]" if $vals[3] and $vals[3] ne $vals[2];
    return $val;
}
```

**Key Features:**
- Handles fixed focal length lenses (same min/max focal)
- Handles fixed aperture lenses (same min/max aperture)
- Special handling for Pentax Q cameras that write 0 for max focal on fixed lenses
- Converts 'inf' and 'undef' values to '?' for unknown values

### ConvertLensInfo Function

**Location:** `WriteExif.pl:73-78`

The inverse function for writing:

```perl
sub ConvertLensInfo($)
{
    my $val = shift;
    my @a = GetLensInfo($val, 1); # allow unknown "?" values
    return @a ? join(' ', @a) : $val;
}
```

Uses `GetLensInfo()` helper (`Exif.pm:5719-5735`) to parse lens strings.

## Special Handling and Edge Cases

### 1. Pentax Quirk
- Pentax Q cameras write 0 for the maximum focal length of fixed focal length lenses
- PrintLensInfo handles this by checking if max focal is 0 and treating it as equal to min focal

### 2. Unknown Values
- Some cameras may write 'inf' or 'undef' for unknown values
- PrintLensInfo converts these to '?' in the output
- The parser accepts '?' as a valid unknown marker

### 3. Variable Aperture Lenses
- Zoom lenses often have different maximum apertures at different focal lengths
- Format: "18-55mm f/3.5-5.6" where f/3.5 is at 18mm and f/5.6 is at 55mm

### 4. DNG Files
- DNG format has its own DNGLensInfo tag (0xC630) with identical structure
- Uses the same PrintLensInfo/ConvertLensInfo functions
- Located in IFD0 instead of ExifIFD

## Related Composite Tags

ExifTool also provides composite tags that derive lens information from multiple sources:

### Lens Composite Tag
**Location:** `Exif.pm:5257-5277`

Attempts to provide lens identification from:
1. LensModel tag (preferred)
2. Lens tag (manufacturer-specific)
3. XMP-aux:LensID
4. Falls back to manufacturer-specific lens lookup tables

### LensID Composite Tag
**Location:** `Exif.pm:5196-5223`

Complex lens identification system that uses:
- LensType from maker notes
- Focal length and aperture information
- Manufacturer-specific lookup tables
- LensModel for disambiguation

## Implementation Notes for exif-oxide

1. **Data Format**: LensInfo stores 4 RATIONAL values (8 bytes each = 32 bytes total)
2. **Validation**: Must validate all 4 values are present for proper formatting
3. **PrintConv**: Essential for user-friendly output - raw rationals are not useful
4. **Edge Cases**: Must handle Pentax Q quirk and unknown value markers
5. **String Encoding**: LensMake/LensModel are ASCII strings with null termination

## Trust ExifTool Principle

The lens tag handling demonstrates why trusting ExifTool is critical:
- Special handling for Pentax Q cameras discovered through real-world testing
- Complex lens identification system built over decades
- Manufacturer-specific quirks encoded in the logic
- Composite tag system that provides fallbacks for missing data