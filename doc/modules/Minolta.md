# Minolta.pm Module Outline

**ExifTool Version:** 13.26
**Module Version:** 2.88
**Document Version:** 1.0
**Last Updated:** 2025-06-30

## Overview

Minolta.pm handles Minolta EXIF maker notes tags and provides foundational support for Sony cameras (which inherited Minolta's camera business). This module is crucial for both legacy Minolta cameras and early Sony DSLR cameras, featuring extensive lens databases, camera-specific settings, and advanced adapter detection for Canon EF and Sigma SA lenses.

## Module Structure

### Tag Tables (11 total)

**Primary Tables:**

1. **Main** - Primary maker notes table (408+ hex tag definitions)
2. **CameraSettings** - General camera settings and parameters
3. **CameraSettings7D** - Minolta 7D specific settings
4. **CameraSettings5D** - Minolta 5D specific settings
5. **CameraInfoA100** - Sony A100 camera information
6. **ISInfoA100** - Sony A100 image stabilization information
7. **CameraSettingsA100** - Sony A100 camera settings
8. **WBInfoA100** - Sony A100 white balance information

**Video/Audio Tables:** 9. **MOV1**, **MOV2** - QuickTime video metadata 10. **MMA** - Audio metadata

### Key Data Structures

- **%minoltaLensTypes**: Comprehensive lens database with 300+ entries including Minolta AF lenses
- **%minoltaTeleconverters**: Teleconverter identification lookup
- **%minoltaColorMode**: Color mode definitions for Minolta cameras
- **%sonyColorMode**: Color mode definitions for Sony cameras
- **%minoltaSceneMode**: Scene mode definitions
- **%afStatusInfo**: AF status interpretation (focus confirmation data)
- **%metabonesID**: Metabones adapter identification for Canon EF lenses

## Processing Architecture

### 1. Main Processing Flow

**Standard EXIF Processing**: Uses standard TIFF-based maker note structure with full write support via `WriteExif` and `CheckExif` procedures.

**Multi-Brand Support**: Seamlessly handles both Minolta and early Sony cameras using the same tag structure with conditional processing.

### 2. Special Processing Functions

- **ConvertWhiteBalance()**: Converts white balance values with special handling for unknown values
- **Lens Adapter Detection**: Advanced logic for detecting Canon EF and Sigma SA lenses via adapters
- **AF Status Processing**: Detailed focus confirmation analysis
- **Model-Specific Conditionals**: Extensive per-model tag processing

## Key Features

### Comprehensive Lens Database

- **Native Minolta AF Lenses**: Complete A-mount lens catalog from 1985-2006
- **Sony Compatibility**: Early Sony A-mount lenses included
- **Third-Party Support**: Extensive Sigma, Tamron, and Tokina lens database
- **Canon EF Adapter Support**: Metabones, Fotodiox, Sigma, and Viltrox adapter detection
- **Sigma SA Adapter Support**: MC-11 SA-E adapter with Sigma SA-mount lenses
- **Teleconverter Detection**: Automatic identification of attached teleconverters

### Advanced Adapter Detection

- **Metabones Adapters**: Smart Adapter, Speed Booster, Speed Booster Ultra
- **Canon EF Lens Identification**: Maintains Canon lens database compatibility
- **Sigma SA-E Detection**: MC-11 adapter with Sigma SA-mount lens support
- **Firmware Version Handling**: Accounts for adapter firmware differences

### Camera-Specific Features

- **Minolta 7D/5D**: Dedicated camera settings tables
- **Sony A100**: Comprehensive camera information, IS, and WB tables
- **Anti-Shake System**: In-body image stabilization information
- **Creative Styles**: Color mode and scene mode processing
- **AF System**: Detailed autofocus status and sensor information

### Multi-Format Support

- **Still Images**: JPEG, MRW (Minolta RAW), ARW (Sony RAW)
- **Video**: MOV containers with Minolta/Sony specific metadata
- **Audio**: MMA audio format support

## Special Patterns

### Lens Adapter Detection Logic

```perl
# Metabones adapter detection
my $id = $val & 0xff00;
my $mb = $metabonesID{$id};
if ($mb) {
    ref $mb or $id = $mb, $mb = $metabonesID{$id};
    require Image::ExifTool::Canon;
    my $lens = $Image::ExifTool::Canon::canonLensTypes{$val - $id};
    return "$lens + $$mb" if $lens;
}
```

### AF Status Processing

```perl
# Focus status with detailed position information
PrintConv => {
    0 => 'In Focus',
    -32768 => 'Out of Focus',
    OTHER => sub {
        return $val < 0 ? "Front Focus ($val)" : "Back Focus (+$val)";
    },
}
```

### Model-Specific Conditionals

```perl
# Different processing for Minolta vs Sony
{
    Condition => '$$self{Make} =~ /^SONY/',
    Name => 'SonySpecificTag',
    # Sony-specific processing
},
{
    Condition => '$$self{Make} !~ /^SONY/',
    Name => 'MinoltaSpecificTag',
    # Minolta-specific processing
}
```

## Usage Notes

### Supported Models

- **Minolta DSLRs**: Dynax/Maxxum 7D, 5D, A1, A2 (DiMAGE)
- **Sony DSLRs**: Early A-mount cameras (A100, A200, A300, A350, A700, A900)
- **Legacy Digital**: DiMAGE series digital cameras
- **Film Cameras**: Some late film cameras with digital backs

### Lens Compatibility

- **A-Mount System**: Full Minolta AF lens compatibility
- **Adapter Support**: Canon EF, Sigma SA via electronic adapters
- **Manual Lenses**: Some support for manual focus lenses with electronic contacts
- **Teleconverter Support**: 1.4x and 2x teleconverters

### Write Capabilities

- Full maker note writing support
- Lens information updates
- Camera settings modifications
- White balance adjustments

## Debugging Tips

### Common Issues

1. **Adapter Detection**: Firmware version differences may affect lens identification
2. **Model Transitions**: Sony cameras may use different tag layouts than Minolta
3. **Lens Database**: Some rare or region-specific lenses may not be identified
4. **AF Status**: Manual focus lenses may not provide meaningful AF status

### Diagnostic Features

- Extensive lens database with fallback options
- AF status provides focus position information
- White balance information includes fine-tuning data
- Scene mode detection for automatic settings

### Adapter-Specific Considerations

- **Metabones Smart Adapter**: Firmware versions before 31 may have compatibility issues
- **Canon EF Identification**: Requires proper Canon lens database loading
- **Sigma SA-E**: Limited to Sigma SA-mount lens database
- **Electronic Contacts**: Adapter quality affects metadata accuracy

## Historical Context

### Minolta Heritage

- Minolta developed the A-mount system in 1985
- Sony acquired Minolta's camera business in 2006
- Early Sony cameras maintained Minolta compatibility

### Technology Evolution

- Anti-shake technology (in-body stabilization)
- Advanced AF systems with position feedback
- Digital integration of film-era lens designs
- Electronic adapter technology for cross-brand compatibility

## References

Extensive documentation from Minolta technical specifications, Sony camera documentation, reverse engineering efforts, and comprehensive testing with various camera models and lens combinations. Multiple private communications with photographers and technical experts contributed to the lens database accuracy.
