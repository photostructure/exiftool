# GPS Coordinate Sign Handling

**ExifTool Version:** 13.26  
**Document Version:** 1.0  
**Last Updated:** 2025-07-31

## Overview

ExifTool's GPS coordinate sign handling is implemented through a two-stage process that combines raw coordinate values with reference direction tags to produce properly signed decimal degrees. Raw GPS coordinates in EXIF are always stored as positive rational values, with separate reference tags indicating hemisphere (N/S for latitude, E/W for longitude).

## Core Algorithm

### 1. Raw GPS Data Structure

**EXIF GPS Tags:**
- `GPSLatitude` - Always positive rational array [degrees, minutes, seconds]
- `GPSLatitudeRef` - String: "N"/"North" or "S"/"South"  
- `GPSLongitude` - Always positive rational array [degrees, minutes, seconds]
- `GPSLongitudeRef` - String: "E"/"East" or "W"/"West"

### 2. Sign Determination Logic

**Location:** `lib/Image/ExifTool/GPS.pm` lines 381, 402

**GPSLatitude Sign Logic:**
```perl
ValueConv => '$val[1] =~ /^S/i ? -$val[0] : $val[0]'
```

**GPSLongitude Sign Logic:**  
```perl
ValueConv => '$val[1] =~ /^W/i ? -$val[0] : $val[0]'
```

**Algorithm:**
1. Extract raw positive coordinate value from GPSLatitude/GPSLongitude
2. Check reference tag (GPSLatitudeRef/GPSLongitudeRef)
3. Apply sign:
   - **Latitude**: Negative if reference starts with "S" (case-insensitive), positive otherwise
   - **Longitude**: Negative if reference starts with "W" (case-insensitive), positive otherwise

### 3. Implementation Details

**Composite Tag Processing:**

The sign application happens in the Composite GPS tags, not the raw EXIF tags:

```perl
# GPSLatitude Composite (lines 368-384)
GPSLatitude => {
    Require => {
        0 => 'GPS:GPSLatitude',      # Raw positive coordinate
        1 => 'GPS:GPSLatitudeRef',   # Direction reference
    },
    ValueConv => '$val[1] =~ /^S/i ? -$val[0] : $val[0]',  # Apply sign
    PrintConv => 'Image::ExifTool::GPS::ToDMS($self, $val, 1, "N")',
}

# GPSLongitude Composite (lines 385-405)  
GPSLongitude => {
    Require => {
        0 => 'GPS:GPSLongitude',     # Raw positive coordinate
        1 => 'GPS:GPSLongitudeRef',  # Direction reference
    },
    ValueConv => '$val[1] =~ /^W/i ? -$val[0] : $val[0]',  # Apply sign
    PrintConv => 'Image::ExifTool::GPS::ToDMS($self, $val, 1, "E")',
}
```

### 4. Reference Direction Processing

**Location:** `lib/Image/ExifTool/GPS.pm` lines 23-49

**Flexible Input Handling:**

ExifTool accepts multiple formats for reference directions when writing:

```perl
# Latitude Reference (lines 23-35)
%printConvLatRef = (
    OTHER => sub {
        my ($val, $inv) = @_;
        return undef unless $inv;
        return uc $2 if $val =~ /(^|[^A-Z])([NS])(orth|outh)?\b/i;  # N/North, S/South
        return $1 eq '-' ? 'S' : 'N' if $val =~ /([-+]?)\d+/;      # Signed numbers
        return undef;
    },
    N => 'North',
    S => 'South',
);

# Longitude Reference (lines 37-49)
%printConvLonRef = (
    OTHER => sub {
        my ($val, $inv) = @_;
        return undef unless $inv;
        return uc $2 if $val =~ /(^|[^A-Z])([EW])(ast|est)?\b/i;   # E/East, W/West
        return $1 eq '-' ? 'W' : 'E' if $val =~ /([-+]?)\d+/;      # Signed numbers
        return undef;
    },
    E => 'East',
    W => 'West',
);
```

**Accepted Input Formats:**
- Full names: "North", "South", "East", "West"
- Abbreviations: "N", "S", "E", "W"  
- Signed numbers: Positive for N/E, negative for S/W
- Case-insensitive matching

## Key Functions

### ToDegrees Function

**Location:** `lib/Image/ExifTool/GPS.pm` lines 582-600

**Purpose:** Converts coordinate strings to decimal degrees with optional sign handling

**Critical Sign Logic (line 598):**
```perl
$deg = -$deg if $doSign ? $val =~ /[^A-Z](S(outh)?|W(est)?)\s*$/i : $deg < 0;
```

**Parameters:**
- `$val` - Input coordinate string
- `$doSign` - Boolean flag to enable S/W sign detection
- `$coord` - Optional 'lat'/'lon' for coordinate pair extraction

**Algorithm:**
1. Extract numeric values from input string using regex
2. Convert DMS to decimal degrees: `degrees + minutes/60 + seconds/3600`
3. Apply negative sign if:
   - `$doSign` is true AND coordinate ends with S/South or W/West
   - OR `$doSign` is false AND value is already negative

### ToDMS Function  

**Location:** `lib/Image/ExifTool/GPS.pm` lines 495-574

**Purpose:** Converts decimal degrees to DMS format with reference directions

**Sign Handling (lines 506-515):**
```perl
if ($val < 0) {
    $val = -$val;
    $ref = {N => 'S', E => 'W'}->{$ref};  # Flip reference direction
    $sign = '-';
    $minus = '-';
}
```

**Algorithm:**
1. Check if input value is negative
2. Make value positive for DMS conversion
3. Flip reference direction: N→S, E→W
4. Store sign information for formatting

## Edge Cases and Special Handling

### 1. Invalid Values
- Returns empty string for "inf" or "undef" values
- Gracefully handles missing or malformed reference tags

### 2. Coordinate Pair Extraction
- Supports combined coordinate strings: "40.4462 N, 73.9865 W"
- Can extract latitude or longitude from combined input
- Uses regex matching for cardinal direction detection

### 3. Scientific Notation
- Supports scientific notation input: "4.04462E+01"
- Handles positive/negative exponents correctly

### 4. Round-off Error Handling
- DMS conversion includes round-off correction (lines 560-563)
- Ensures minutes and seconds stay below 60
- Example: "72° 59' 60.00"" → "73° 0' 0.00""

## Implementation Notes

### For exif-oxide Implementation

**Key Points:**
1. **Two-stage process**: Raw EXIF tags are always positive, Composite tags apply signs
2. **Reference-based signs**: Never trust coordinate sign directly, always check reference tags
3. **Case-insensitive matching**: Reference detection uses `/^S/i` and `/^W/i` patterns
4. **Flexible input**: Support multiple reference formats during writing
5. **Round-off handling**: Include precision correction for DMS boundary cases

**Critical Regex Patterns:**
- Latitude: `$val[1] =~ /^S/i ? -$val[0] : $val[0]`
- Longitude: `$val[1] =~ /^W/i ? -$val[0] : $val[0]`

**File References:**
- Main logic: `/lib/Image/ExifTool/GPS.pm` lines 368-405 (Composite tags)
- Conversion functions: `/lib/Image/ExifTool/GPS.pm` lines 495-600
- Reference handling: `/lib/Image/ExifTool/GPS.pm` lines 23-49