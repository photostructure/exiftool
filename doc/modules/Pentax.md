# Pentax.pm Module Outline

**ExifTool Version:** 13.26  
**Module Version:** 3.48  
**Document Version:** 1.0  
**Last Updated:** 2025-06-29

## Overview

The Pentax module handles maker notes for Pentax/Asahi cameras and their Samsung rebadges. It features extensive lens identification (700+ lenses), encrypted data decoding, and comprehensive support for camera settings across multiple generations of Pentax DSLRs and mirrorless cameras.

## Module Structure

### Tag Tables (49 total)

**Primary Tables:**
- **%Pentax::Main** - Primary maker notes dispatcher
- **%pentaxLensTypes** - Lens identification database (700+ entries)
- **%Pentax::CameraSettings** - Core camera configuration
- **%Pentax::ShotInfo** - Shot-specific information

**Camera-Specific Tables:**
- **%Pentax::Type2-Type10** - Generation-specific formats
- **%Pentax::KP** - Modern mirrorless/DSLR format
- Various model-specific tables for K-1, K-3, K-5, K-70, etc.

**Feature Tables:**
- **%Pentax::AFInfo** - Autofocus information
- **%Pentax::FlashInfo** - Flash settings
- **%Pentax::BatteryInfo** - Battery status
- **%Pentax::LensInfo** - Lens data variants

### Lens Database

**Lens Identification System:**
- Two-byte format: Series + Model
- Series codes: K=1, A=2, F=3, FA=3-6, DA=3,4,7, DFA=4,7, etc.
- 700+ lens definitions
- Third-party support (Sigma, Tamron, Tokina)

**Special Handling:**
- Firmware version variations
- SDM lens detection
- Series number corrections
- Legacy lens compatibility

## Processing Architecture

### 1. Format Detection
```perl
ProcessPentax($$$)
  ├── Detect Asahi vs Pentax
  ├── Determine format type (Type2-Type10, KP)
  ├── Handle encrypted data
  └── Process subdirectories
```

### 2. Encryption System

**CryptShutterCount($$)**
- Decrypts shutter count data
- Camera-specific algorithms
- Obfuscation techniques

**Encrypted Tags:**
- Shutter count
- Various camera settings
- Model-specific data

### 3. Special Processing

**DecodeAFPoints($$$$)**
- AF point bitmap decoding
- Model-specific layouts
- Active point detection
- Area mode support

**PrintFilter($$$)**
- Digital filter decoding
- Effect parameters
- Custom processing

## Key Features

### AF System Support

**AF Types:**
- SAFOX VIII, IX, X, XI
- Different point counts
- Phase detection layouts
- Contrast detection areas

**AF Data:**
- Selected points
- Focus mode
- Tracking information
- Fine adjustment values

### Shake Reduction

**SR System:**
- On/off status
- Focal length setting
- Effectiveness data
- SR timing modes

### Custom Functions

**Model-Specific:**
- Different sets per camera
- Firmware variations
- Menu customization
- Button assignments

### Time/GPS Features

**World Time:**
- Home/destination cities
- DST settings
- Time zone data

**GPS Integration:**
- Built-in GPS support
- Track logs
- Geotagging options

## Camera Evolution

### Format Types

**Type 2** (Early DSLRs)
- *ist D series
- Basic maker notes

**Type 3** (Mid-era)
- K10D, K100D
- Enhanced features

**Type 4-6** (Modern DSLRs)
- K-7, K-5, K-3 series
- Advanced data structures

**Type 7-10** (Recent)
- K-1, K-70
- Latest features

**KP Format** (Current)
- K-3 III, K-1 II
- Unified structure

### Feature Progression

**Early Models:**
- Basic exposure data
- Simple AF systems
- Limited customization

**Modern Models:**
- Pixel Shift Resolution
- Advanced SR
- Astrotracer
- Electronic shutter

## Special Patterns

### Lens Detection
```perl
# Series correction for older firmware
$val =~ s/^4 /7 / and $$conv{$val} and return "$$conv{$val} ($_[0])";
```

### Encryption
```perl
# Model-specific decryption
my $key = $camera_key{$model} || $default_key;
```

### AF Points
```perl
# Bitmap to point names
my @points = DecodeAFPoints($val, $model, $afInfo, $rotated);
```

## Unique Features

### Pixel Shift
- Multi-shot high resolution
- Motion correction
- Raw file handling

### Astrotracer
- Star tracking mode
- GPS integration
- Exposure calculation

### Digital Filters
- In-camera effects
- Parameter storage
- Preview generation

### Weather Sealing
- Environmental data
- Temperature readings
- Pressure sensors (some models)

## Brand Relationships

### Samsung Partnership
- GX series rebadges
- Shared maker notes
- Lens compatibility

### Ricoh Acquisition
- Continued Pentax brand
- Technology integration
- Format consistency

## Usage Notes

- Extensive lens database
- Encrypted data common
- Model-specific variations
- Good third-party lens support
- Active development continues

## Debugging Tips

- Check format type first
- Encryption varies by model
- AF system changes frequently
- Lens ID corrections may apply
- Watch for Samsung variants