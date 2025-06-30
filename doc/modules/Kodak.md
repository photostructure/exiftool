# Kodak.pm Module Outline

**ExifTool Version:** 13.26  
**Module Version:** 1.48  
**Document Version:** 1.0  
**Last Updated:** 2025-06-30

## Overview

The Kodak module handles metadata extraction from Kodak cameras, spanning from early digital cameras to modern video cameras. It's notable for having the most diverse and inconsistent maker note formats of any camera manufacturer, with 11 different maker note types plus extensive video metadata support. The module covers over 100 different Kodak camera models from the DC50 era through modern PixPro cameras.

## Module Structure

### Tag Tables (27 total)

**Core Maker Note Types:**

- `Main` (Type 1) - Most common format (C360, C663, CX7330, DX4900, Z650, etc.)
- `Type2` - Early models (DC220, DC260, DC265, DC290) + HP/Pentax/Minolta rebrands
- `Type3` - Mid-range models (DC240, DC280, DC3400, DC5000)
- `Type4` - Basic models (DC200, DC215)
- `Type5` - CX4200 series (CX4200, CX4210, CX4230, CX4300, CX4310, CX6200, CX6230)
- `Type6` - DX3215, DX3700
- `Type7` - Serial number-based (C340, C433, CC533, LS755, V803, V1003)
- `Type8` - IFD-format maker notes (ZD710, P712, P850, P880, V1233, etc.)
- `Type9` - Compact models (C140, C180, C913, C1013, M320, M340, M550)
- `Type10` - Z980 (IFD with byte order indicator)
- `Type11` - PixPro S-1 (similar to Ricoh/GE formats)

**SubIFD Architecture:**

- `SubIFD0` through `SubIFD6` - Nested IFDs for newer models
- `CameraInfo` - P712, P850, P880 specific camera information
- `Unknown` - Fallback for unrecognized maker notes

**Legacy Formats:**

- `IFD` - Separate IFD in older JPEG/TIFF/DCR/KDC images (DC50, DC120, DCS760C, Pro Back)
- `KDC_IFD` - KDC images from newer models (P880, Z1015IS)
- `TextualInfo` - Text-based metadata format
- `Processing` - White balance adjustment data

**APP3 "Meta" Segment:**

- `Meta` - APP3 "Meta" segment tags (DC280, DC3400, DC5000, MC3, M580, Z950, Z981)
- `SpecialEffects` - Special effects sub-IFD
- `Borders` - Borders sub-IFD

**Video Metadata:**

- `MOV` - MOV video tags (P880 format)
- `DcMD`, `DcME`, `DcEM` - MOV/MP4 metadata atoms
- `Free` - M5370 MP4 "free" atom (bad form - useful data in unused space)
- `frea` - PixPro SP360 MP4 "frea" atom
- `Scrn` - Preview info in MP4 free/Scrn atom
- `pose` - Accelerometer data from PixPro 4KVR360

**Composite:**

- `Composite` - Derived tags (DateCreated, WB_RGBLevels calculations)

### Key Data Structures

**Constants:**

- `%sceneModeUsed` - Scene mode lookup table for SubIFD2/SubIFD3
- Various PrintConv hashes for tag value interpretation

**Complex Structures:**

- Multiple SubIFD pointer arrays in Type8 (0xfc00-0xfc06, 0xfcff, 0xff00)
- Extensive white balance matrix tables in IFD format
- Professional camera calibration data (DCS series)
- Color matrix tables for different lighting conditions

## Processing Architecture

### 1. Main Processing Flow

The module uses standard ExifTool processing with format-specific dispatching:

- Binary data processing for most maker note types
- IFD processing for Type8, Type10, and legacy IFD formats
- Text processing for TextualInfo format
- Specialized processing for video metadata

### 2. Special Processing Functions

**ProcessKodakIFD($$$)**

- Handles IFDs with leading byte order mark (Type8 SubIFDs)
- Validates and sets byte order from 2-byte prefix
- Delegates to standard Exif processing after setup

**ProcessKodakText($$$)**

- Parses colon-separated text metadata
- Dynamic tag name generation for unknown fields
- Handles multi-line format with automatic tag creation

**ProcessPose($$$)**

- Extracts accelerometer and angular velocity data from PixPro 4KVR360
- Supports streamed orientation data extraction
- Uses ExtractEmbedded option for full data extraction

**WriteKodakIFD($$$)**

- Writes IFDs with byte order mark preservation
- Handles fixup adjustments for byte order mark length
- Maintains compatibility with ProcessKodakIFD

**CalculateRGBLevels(@)**

- Complex white balance calculation using multiple input tags
- Combines white balance multipliers, coefficients, and color temperature
- Performs polynomial calculation for camera-specific RGB levels

## Key Features

### 1. Format Diversity Management

**Inconsistent Maker Notes Strategy:**

- 11 different binary formats plus text and IFD formats
- Model-specific conditional processing
- Graceful degradation for unknown formats

**Legacy Support:**

- Extensive support for 1990s-2000s digital cameras
- Professional camera support (DCS series, Pro Back)
- Film scanner metadata (some models)

### 2. Advanced Metadata Features

**Professional Camera Support:**

- Color matrix tables for different lighting conditions
- Extensive calibration data (linearity, tone curves, noise reduction)
- Sensor characterization data
- Image processing parameter tables

**Video Metadata:**

- MOV and MP4 format support
- Accelerometer data extraction
- Preview image handling
- Streaming orientation data

**White Balance System:**

- Multiple WB calculation methods
- Color temperature integration
- RGB coefficient processing
- Composite tag generation

### 3. Special Processing Patterns

**Conditional Tag Processing:**

- Make/model-specific tag interpretation (Type9 FNumber scaling)
- Camera-specific serial number formats
- Conditional SubIFD processing

**Binary Data Handling:**

- Extensive use of Binary => 1 for large data blocks
- Custom format specifications for complex data
- FixCount flags for problematic tag counts

## Special Patterns

### 1. SubIFD Architecture (Type8)

Type8 uses a complex nested SubIFD structure:

- Main IFD contains pointers to 7 SubIFDs (0xfc00-0xfc06, 0xfcff)
- Each SubIFD can be either 'undef' format or standard SubIFD pointer
- Different base offset calculations for different SubIFDs
- Conditional processing based on pointer values

### 2. Format Evolution

**Early Cameras (Types 2-7):**

- Simple binary data formats
- Fixed-length structures
- Basic camera settings

**Mid-Period (Type8):**

- Complex IFD structure
- Extensive professional features
- Multiple calibration tables

**Modern Cameras (Type9-11):**

- Simplified formats
- Video integration
- Cross-manufacturer compatibility

### 3. Text Processing Innovation

The TextualInfo processing demonstrates sophisticated text parsing:

- Automatic tag name generation from descriptive text
- Flexible key-value pair extraction
- Graceful handling of unstructured data

## Usage Notes

### Model Identification

**Format Detection:**

- No explicit format indicator - relies on data pattern recognition
- Model names in NOTES sections indicate supported cameras
- Fallback to Unknown table for unrecognized formats

**Camera Coverage:**

- Consumer: C-series, CX-series, DX-series, LS-series, V-series, Z-series
- Professional: DCS series, Pro Back series
- Video: PixPro series, Playsport series
- Rebranded: HP PhotoSmart, Pentax EI-series, Minolta EX1500Z

### White Balance Handling

**Multiple Calculation Methods:**

- Simple RGB levels from KDC_IFD
- Complex calculation from KodakIFD using CalculateRGBLevels
- Fallback to existing software levels when available

**Professional Features:**

- Multiple lighting condition matrices (Daylight, Tungsten, Fluorescent, Flash)
- Color temperature integration
- Custom white balance support

## Debugging Tips

### Common Issues

**Maker Note Recognition:**

- Use -v3 to see which Type table is being used
- Check NOTES sections for camera model compatibility
- Unknown format indicates unsupported camera or corrupted data

**SubIFD Processing:**

- Type8 SubIFDs may have zero pointers (normal)
- FixCount flags indicate known problematic cameras
- Base offset differences can cause extraction issues

**Video Metadata:**

- MOV/MP4 tags require specific atom structure
- PixPro accelerometer data needs ExtractEmbedded option
- "free" atom usage is non-standard but intentional

### Professional Camera Debug

**Color Matrix Issues:**

- Check for Standard/Deviant/Unique matrix variants
- Verify lighting condition-specific matrices
- Look for obsolete vs. current matrix tags

**Calibration Data:**

- Many calibration tags are binary data blocks
- Professional processing requires multiple related tags
- Firmware version affects available calibration data

The Kodak module exemplifies handling manufacturer inconsistency while maintaining comprehensive feature support across decades of camera evolution.
